前提：容器配置文件的修改
----------------------

从dockerhub上下载的镜像基本上是不符合环境需求的，启动容器后还需要对配置文件进行修改，修改配置后重启容器，这个过程就显得过于繁琐。当然，也可以使用自制镜像或存储卷的方式修改或替换配置文件，但无法用于批量操作
```diff
- docker在配置文件上的解决方案是，构建镜像时，以变量的方式修改配置文件，启动容器时，docker会使用一个其他程序先替换容器的主进程，这个程序会将用户的传参替换到配置文件中的变量，传参完后，这个程序会通过exec命令再次启动容器的主进程
- 如果用户没有传参，则使用镜像配置文件的默认参数
```
以nginx镜像为例，构建nginx镜像时将nginx的配置文件以变量的方式保存为只读层：
```shell
/etc/nginx/conf.d/server.conf
{
	server_name $NGX_SERVER_NAME;
	listen $NGX_IP:$NGINX_PORT;
	root $DOC_ROOT;
}
```
通过nginx镜像启动容器时，docker会使用一个其他程序读取用户为nginx的配置文件传入的参数，然后保存为新的配置文件，并通过此配置文件启动容器

Dockerfile
==========

Dockerfile用户构建镜像，是一个包含构建指令的文本文件，"#"号开头表示注释，在整个Dockerfile文件中第一个非注释行必须以"FROM"开头，"FROM"将指定一个基础镜像，后续构建的镜像将基于这个基础镜像实现 <br />
注：
- 制作Dockerfile时必须在一个特定的工作目录下进行，且建议Dockerfile文件名首字母大写
- Dockerfile文件中可以引用同级目录或子目录下的文件，但不能引用父目录的文件
- 在工作目录下可以创建一个.dockeringore，写在此隐藏文件中的路径可以作为Dockerfile的黑名单，不被Dockefile引用
- 在Dockerfile中也可以执行shell命令，但这些shell命令必须是引用的底层镜像所包含的

构建指令集
---------

#### FROM
* FROM指令是最重要的一个且必须为Dockerfile文件第一个非注释行，用于为镜像文件构建过程中指定基础镜像，后续的指令行基于此镜像提供的运行环境。默认情况下，docker build会在docker主机上查找指定的基础镜像文件，主机上不存在时会从dockerhub下载基础镜像 
* 语法格式：
```shell
FROM <repository>:[:<tag>] 或
FROM <repository>@<digest>
	digest：镜像的hash码
	repository：默认使用dockerhub的镜像，也可以指定了别的registry
```

#### MAINTAINER（已废弃，被兼容）
* 用于让Dockerfile制作者提供本人信息，Dockerfile不限制MAINTAINER出现的位置，但推荐将其放在FROM指令之后 
* 语法格式：
```shell
MAINTAINER "hebor <hebo1248@163.com>"
```

#### LABEL
* LABEL为镜像指定元数据，LABEL使用键值对的表现形式，MAINTAINER可以通过LABEL实现
* 语法格式：
```shell
LABEL <key>=<value> <key>=<value> ...
LABEL maintainer="hebo1248@163.com"
```

#### COPY
* 用于从Docker主机复制文件到构建的新镜像文件中
* 文件复制准则
	被复制的文件必须与Dockerfile处于同级或下级目录，不能是父目录中的文件
	如果被复制的是目录，则此目录下的所有文件及目录都会被复制，但此目录本身不会被复制
	如果指定了多个源文件或目录，或者在其中使用的通配符，那目标路径必须是一个目录，且必须以"/"结尾
	如果目标路径事先不存在，他将会被自动创建，包括父目录路径
* 语法格式
```shell
COPY <src> ... <dest> 或
COPY ["<src>",... "<dest>"]    #通常源文件路径中存在空白字符时使用第二种格式
```

补充：快速测试容器命令
```shell
# docker build -t test:v0.1 /srv/dockerfile/    
	给镜像打标签，构建镜像时必须指定工作目录

# docker run --rm test:v0.1 cat /data/www/html/index.html
	启动容器，仅执行cat命令，退出删除容器
```

#### ADD
* ADD指令类似COPY指令，ADD支持使用tar文件和url路径，此指令似乎不支持使用"&&"语法
* 操作准则：
	COPY的文件复制准则在ADD中一样适用
	如果源为url且目标路径不是目录，则源指定的url将被下载并直接重命名为目标文件；如果目标路径是目录，则源指定的url文件将会被下载到目标目录下
	如果源是一个压缩文件，它将会被展开为解压后的目录；通过url获取到的压缩文件则不会自动展开
	有多个源目标或使用通配符时，目标路径必须是个目录，如果目标路径不以"/"结尾，则所有源文件的内容都会被直接写入目标文件
* 语法格式：
```shell
ADD <src> ... <dest> 或
ADD ["<src>",... "<dest>"]
```

#### WORKDIR
* 为Dockerfile中所有的RUN、CMD、ENTRYPOINT、COPY和ADD指定工作目录。在Dockerfile中，WORKDIR指令可出现多次，其路径也可以为相对路径，不过相对路径是作用在此前一个WORKDIR指令指定的路径之上的
* 语法格式
```shell
WORKDIR $PATH
WORKDIR /var/log
```

#### VOLUME
* 用于在镜像中创建一个挂载卷，但在Dockerfile中只能使用docker-managed volume
* 语法格式
```shell
VOLUME <mountpoint> 或
VOLUME ["<mountpoint>"]
```

#### EXPOSE
* 用于为容器打开指定要监听的端口，EXPOSE只能指定容器要开放的端口，使用随机高端口的方式映射到宿主机，不能直接指定映射到宿主机的端口。默认启动容器时指定的端口不会暴露，需要通过-P选项手动暴露端口
* 语法格式
```shell
EXPOSE <port>[/<protocol>] [<port>[/<protocol>] ...]
EXPOSE 11211/udp 11211/tcp
```

#### ENV
* 用于为镜像定义所需要的环境变量，可被Dockerfile文件后续的其他指令调用，调用格式为$variable_name或${variable_name}
* 语法格式
```shell
ENV <key> <value> 或    #此格式中，<key>之后的所有内容都被视为<value>的组成，所以一次只能设置一个变量
ENV "<key>=<value>"    #此格式可一次设置多个变量，每个变量通过键值对的方式表现为"<key>=<value>"，如果<value>中包含空格，可以用\转义或对其加引号进行标识，反斜线也可用于换行
```

补充：docker run和docker build的区别
镜像构建和启动容器是两个不同的过程，如果在Dockerfile文件中设置ENV环境变量，在docker run时能否执行变量替换
```shell
Dockerfile  docker build构建镜像             docker run启动容器
文件       ----------------------> 镜像文件 --------------------> 容器
```
解析：Dockerfile中定义的所有环境变量是容器启动以后可以直接在容器中引用的变量，docker build的过程中已经将Dockerfile中的传参做成了只读镜像，启动容器时可以向容器内传参以修改环境变量，但docker run传参只是显示环境变量已修改，并不会修改docker build的值 <br />
docker build的过程中已经将环境变量的值做成了只读镜像，docker run只是在容器的读写层修改了环境变量，并覆盖了只读镜像的环境变量

#### RUN
* 用于指定**docker build过程中**运行的程序，可以是任何命令，但使用的命令在基础镜像中必须存在，RUN命令也可以运行多次，建议使用换行符一行执行多条命令
* 语法格式
```shell
RUN <command> 或
RUN ["<executable>","<param1>","<param2>"]
```
* 第一种格式中，<command>通常是一个shell命令，以"/bin/sh -c"来运行一个进程，这意味着此进程在容器中PID不为1，不能接收Unix信号，因此，当使用docker stop <container>命令停止容器时，此进程接收不到SIGTERM信号
* 第二种语法格式的参数是一个json数组，<executable>是要运行的命令，<paramN>作为传递给命令的选项或参数；这种格式的命令不会以"/bin/sh -c"来发起，因此常见的shell操作如变量替换以及通配符替换将不会运行

补充：在Linux系统下运行一个服务或程序时，其父进程一定会是shell，所有进程在被终止时一定会将其下的所有子进程一并终止，shell也是如此。而想要不被终止则只能绕过shell，直接通过内核启动程序(nohub command)，直接通过内核启动的程序不会自动释放内存，也不会随着shell的终止而自动终止，但通过内核直接启动程序却不能使用通配符、管道、重定向等功能，因为这些功能是由shell提供的特性 <br />
回到容器，docker的核心理念是一个容器只运行一个进程。而这个进程是直接通过内核启动还是通过shell托管就非常关键，通过容器启动一个进程时，这个进程在整个namespaces中进程号为1，也就是说这个进程由内核启动，所以在命令上就不能使用shell特性，Dockerfile中的RUN命令中的两种语法格式就代表两种启动程序的方式 <br />
启动容器时，如果既想以shell的方式启动进程，又不想shell作为pid号为1的进程，那就需要使用exec启动新的进程来覆盖掉shell进程。使用exec进入已启动的容器后再使用ps命令查看进程时，shell进程的pid依旧为1，这是为了确保容器能够自动接收UNIX信号，但从使用exec命令进入容器的那一刻起，就说明exec命令确实已经替换掉了容器中的主进程

#### CMD
* 用于指定**docker run过程中**运行的命令，即启动容器时默认要运行的程序，容器默认只运行一个程序，所以虽然CMD可以出现多次，但只有最后一个生效。CMD指定的命令可以被docker run的命令行覆盖
* 语法格式
```shell
CMD <command> 或
CMD ["<executable>","<param1>","<param2>"] 或	#前两种语法格式要求与CMD相同
CMD ["<param1>","<param2>"]		#此语法则用于为ENTRYPOINT指令提供默认参数
```

#### ENTRYPOINT
* 类似CMD指令的功能，为容器指定默认运行程序，但ENTRYPOINT启动的程序不会被docker run命令行指定的参数覆盖，但这些**命令行参数**会覆盖CMD指令的内容，并会附加到ENTRYPOINT指定的应用程序后作为其**参数**使用。不过docker run命令的--entrypoint选项可覆盖ENTRYPOINT指令指定的程序
* 语法格式
```shell
ENTRYPOINT <command>
ENTRYPOINT ["<executable>","<param1>","<param2>"]
```

#### USER
* 用于指定运行镜像时或运行Dockerfile中任何RUN、CMD或ENTRYPOINT指令指定的程序时的用户名或UID。默认容器运行身份为root
* 语法格式
```shell
USER <UID>|<UserName>	#UID必须是/etc/passwd中已存在的用户
```

#### HEALTHCHECK
* docker判断容器正常与否并不是判断容器内进程是否正常提供服务，而是仅判断容器进程是否运行。这种判定机制并不能真正的判断容器是否正常，此时需要用一些工具去测试服务是否正常，比如curl、wget。HEALTHCHECK指令用于指定一条CMD命令，这条命令用于检查主进程服务状态
* 语法格式
```shell
HEALTHCHECK [OPTIONS] CMD command	#CMD是关键词，不可少
HEALTHCHECK NONE	#不使用健康检查，会关闭默认的健康检查
```
HEALTHCHECK属于周期性任务，所以会有OPTIONS：
```shell
--interval=DURATION(default:30s)	#检测间隔时间
--timeout=DURATION(default:30s)		#服务未响应超时时间
--start-period=DURATION(default:0s)	#第一次检测延时时间
--retries=N(default:3)				#检测次数
```

#### SHELL
* 用于指定运行程序使用的shell，Linux默认shell为["/bin/sh","-c"]，Windows默认shell为["cmd","\S","\C"]

#### STOPSIGNAL
* 容器内PID为1的进程可以接收Linux的信号，默认信号是15，也就是终止程序。此信号可修改，比如修改为9，修改后docker stop容器进程收到的信号就是强制终止
* 语法格式
```shell
STOPSIGNAL signal	#修改容器主进程可以接收的信号
```

#### ARG
* 定义只在build的过程中使用的变量，且能在执行docker build命令时，通过选项--build-arg <varname>=<value>向biuld过程中传参
* 语法格式
```shell
ARG <name>[=<default value>]	#此处可以不定义默认值，docker build时传参即可
```

#### ONBUILD
* 在Dockerfile中定义一个触发器，自己在docker build基于此Dockerfile制作的镜像时没有问题，但制作的镜像如果被他人FROM引用做基础镜像时，在他人docker build的过程中会执行此触发器
* 语法格式
```shell
ONBUILD <INSTRUCTION>	#INSTRUCTION可以是RUN、CMD或ADD等
ONBUILD ADD http://mirrors.aliyun.com/repo/Centos-7.repo
```
* 几乎任何指令都可以成为触发器指令，但ONBUILD不能自我嵌套，且不会出发FROM和MAINTAINER指令，使用包含ONBUILD指令的Dockerfile构建的镜像应该使用特殊的标签进行声明。在ONBUILD指令中使用ADD或COPY指令时应注意，新构建过程的上下文在缺少指定的源文件时会失败

```diff
- 构建Dockerfile文件时应尽量减少文件行数，因为构建镜像时每一行就是一层镜像，过多的行数会导致镜像臃肿
```

### 向容器中传参
示例：Dockerfile文件内容
```shell
FROM nginx:1.14-alpine
LABEL maintainer="hebor hebo1248@163.com"
ENV NGX_DOC_ROOT="/data/web/html/"			#指定nginx的网页文件目录
EXPOSE 80									#开放80端口
ADD index.html ${NGX_DOC_ROOT}
ADD entrypoint.sh /bin/						#添加nginx的配置文件脚本
CMD ["/usr/sbin/nginx","-g","daemon off;"]	#设置nginx前台运行
ENTRYPOINT ["/bin/entrypoint.sh"]			#执行nginx配置文件脚本
```
示例：entrypoint.sh脚本内容
```shell
#!/bin/sh
cat > /etc/nginx/conf.d/www.conf << EOF
server {
	server_name $HOSTNAME;
	listen ${IP:-0.0.0.0}:${PORT:-80};		#指定默认IP和端口
	root ${NGX_DOC_ROOT:-/usr/share/nginx/html};
}
EOF				#以上都是nginx配置

exec "$@"		#将CMD的参数作为命令，用exec运行，如果docker run的过程中指定了其他参数，则exec会运行docker run命令行参数
```